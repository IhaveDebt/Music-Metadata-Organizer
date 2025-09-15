package main

import (
	"flag"
	"fmt"
	"io"
	"os"
	"path/filepath"
	"strings"

	"github.com/dhowden/tag"
)

func main() {
	dir := flag.String("dir", ".", "music folder")
	dry := flag.Bool("dry", true, "dry run (no moves) by default")
	flag.Parse()

	fmt.Println("Scanning:", *dir)
	filepath.Walk(*dir, func(path string, info os.FileInfo, err error) error {
		if err != nil || info.IsDir() {
			return nil
		}
		ext := strings.ToLower(filepath.Ext(path))
		if ext != ".mp3" && ext != ".flac" && ext != ".m4a" && ext != ".wav" {
			return nil
		}
		f, err := os.Open(path)
		if err != nil {
			fmt.Println("open error:", err)
			return nil
		}
		defer f.Close()
		m, err := tag.ReadFrom(f)
		if err != nil && err != io.EOF {
			fmt.Println("tag read error:", path, err)
			return nil
		}
		artist := m.Artist()
		album := m.Album()
		if artist == "" {
			artist = "Unknown Artist"
		}
		if album == "" {
			album = "Unknown Album"
		}
		targetDir := filepath.Join(*dir, sanitize(artist), sanitize(album))
		targetPath := filepath.Join(targetDir, filepath.Base(path))
		fmt.Printf("File: %s => %s\n", path, targetPath)
		if !*dry {
			os.MkdirAll(targetDir, 0o755)
			os.Rename(path, targetPath)
		}
		return nil
	})
	fmt.Println("Done. Use -dry=false to move files.")
}

func sanitize(s string) string {
	return strings.ReplaceAll(s, "/", "_")
}
